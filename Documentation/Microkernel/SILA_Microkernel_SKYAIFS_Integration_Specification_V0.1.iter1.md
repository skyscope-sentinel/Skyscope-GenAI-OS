# SILA Microkernel & SKYAIFS Integration Specification - V0.1 - Iteration 1

**Based on:** `Documentation/Microkernel/SILA_Microkernel_SKYAIFS_Integration_Specification_V0.1.md` (initial conceptual placeholder from Stage 3 Task Block 1 assignment)
**Key References:**
*   `Documentation/SILA_Language/SILA_Specification_V0.2.md`
*   `Documentation/SKYAIFS_Filesystem/SKYAIFS_Framework_V0.2.md`
*   `Documentation/Microkernel/SILA_Microkernel_Internals_Design_V0.1.md`

**Iteration Focus:** Initial detailing of key SILA IPC protocols for Raw Block I/O, identifying shared I/O buffer structures and capabilities, and outlining an atomicity challenge.

## 1. Introduction
This document begins the detailed specification of the SILA-based interactions between the Skyscope OS Microkernel and SKYAIFS. All designs adhere to SILA V0.2 principles (as defined in `Documentation/SILA_Language/SILA_Specification_V0.2.md`) and leverage its PQC-aware (min 4096-bit security equivalent), verifiable constructs. The goal is to define secure, reliable, and efficient communication pathways.

## 2. Detailed SILA IPC Protocols

### 2.1. Raw Block I/O Protocol (Conceptual SILA V0.2)

This protocol handles requests from SKYAIFS (typically an I/O Path ASA as described in `Documentation/SKYAIFS_Filesystem/SKYAIFS_Framework_V0.2.md`) to the Microkernel's storage service abstraction layer to read or write raw blocks from/to a storage device. SKYAIFS must possess a Microkernel-granted `SILA_CapToken<SILA_RawStorageDevice_Object_Type>` for the target device.

#### 2.1.1. `SKYAIFS_Request_BlockRead_SILA_Call`
*   **Initiator:** SKYAIFS I/O Path ASA.
*   **Target:** Microkernel Storage Service Endpoint (a SILA Capability, e.g., `mkernel_storage_ep_cap`).
*   **SILA Message Structure (`SILA_BlockRead_Request_Record` - a SILA Record type):**
    `{
      request_id: SILA_UniqueID_String, // For tracking asynchronous operations
      storage_device_access_cap: SILA_CapToken<SILA_RawStorageDevice_Object_Type>, // Capability granting access to the specific storage device/partition
      block_id_on_device_lba: SILA_LBA_Integer_Type, // Logical Block Address on the physical device, using a verifiable integer type
      target_buffer_mem_cap: SILA_CapToken<SILA_Memory_Region_Type>, // SKYAIFS-provided memory region capability for read data (must grant write access to Microkernel)
      expected_data_pqc_hash_opt: SILA_Optional<SILA_PQC_Hash_Record<SHA3_512_SILA_Enum>>, // Optional, for read-verify by kernel if supported by device/policy
      reply_to_skyaifs_ipc_ep_cap: SILA_CapToken<SILA_IPC_Endpoint_Type> // SKYAIFS endpoint for completion/error event
    }`
    *   All fields utilize SILA V0.2 primitive types (like `SILA_UniqueID_String`, `SILA_LBA_Integer_Type`) or PQC-aware types (like `SILA_PQC_Hash_Record`). The `target_buffer_mem_cap` is crucial for security and direct data transfer.

#### 2.1.2. `Microkernel_BlockRead_Complete_SILA_Event` (Asynchronous Reply)
*   **Initiator:** Microkernel Storage Service ASA.
*   **Target:** The `reply_to_skyaifs_ipc_ep_cap` specified in the request.
*   **SILA Message Structure (`SILA_BlockRead_Response_Record` - a SILA Record type):**
    `{
      original_request_id_ref: SILA_UniqueID_String, // Correlates with the request_id
      io_status_enum: SILA_IO_OperationStatus_Enum { Success_EnumVal, DeviceError_EnumVal, CapabilityError_EnumVal, PQC_HashMismatchError_EnumVal, ReadOnlyMediaError_EnumVal },
      bytes_actually_read_int: SILA_Verifiable_Integer,
      computed_data_pqc_hash_opt: SILA_Optional<SILA_PQC_Hash_Record<SHA3_512_SILA_Enum>>, // If computed by kernel/device during read
      error_details_sila_record_opt: SILA_Optional<SILA_CapToken<SILA_Error_Record>> // Capability to a detailed error record if status is not Success
    }`
    *   Upon `Success_EnumVal`, the memory region referenced by `target_buffer_mem_cap` (from the request) is populated by the Microkernel.

#### 2.1.3. `SKYAIFS_Request_BlockWrite_SILA_Call`
*   **SILA Message Structure (`SILA_BlockWrite_Request_Record` - a SILA Record type):**
    `{
      request_id: SILA_UniqueID_String,
      storage_device_access_cap: SILA_CapToken<SILA_RawStorageDevice_Object_Type>,
      block_id_on_device_lba: SILA_LBA_Integer_Type,
      source_buffer_mem_cap: SILA_CapToken<SILA_Memory_Region_Type>, // SKYAIFS-provided memory region with data to write (must grant read access to Microkernel)
      pqc_write_integrity_signature_opt: SILA_Optional<SILA_PQC_Signature_Record<MLDSA_5_SILA_Enum>>, // Optional, if blocks are self-signed by SKYAIFS before sending to Microkernel for raw write
      reply_to_skyaifs_ipc_ep_cap: SILA_CapToken<SILA_IPC_Endpoint_Type>
    }`

#### 2.1.4. `Microkernel_BlockWrite_Complete_SILA_Event` (Asynchronous Reply)
*   **SILA Message Structure (`SILA_BlockWrite_Response_Record` - a SILA Record type):**
    `{
      original_request_id_ref: SILA_UniqueID_String,
      io_status_enum: SILA_IO_OperationStatus_Enum { Success_EnumVal, DeviceError_EnumVal, CapabilityError_EnumVal, PQC_IntegrityVerificationError_AtDevice_EnumVal, WriteProtectError_EnumVal },
      bytes_actually_written_int: SILA_Verifiable_Integer,
      error_details_sila_record_opt: SILA_Optional<SILA_CapToken<SILA_Error_Record>>
    }`

## 3. Shared SILA Data Structures & Capability Management

### 3.1. I/O Buffer Management
*   **`SILA_Shared_IO_Buffer_Record` (Conceptual - managed by SKYAIFS or a common Buffer Manager SILA ASA):**
    `SILA_Shared_IO_Buffer_Record {
      buffer_instance_id: SILA_UniqueID_String,
      memory_region_main_cap: SILA_CapToken<SILA_Memory_Region_Type>, // Capability to actual PQC-aware memory allocated by Microkernel
      total_size_bytes_int: SILA_Verifiable_Integer,
      current_data_size_bytes_int: SILA_Verifiable_Integer, // For writes, how much data is in it; for reads, how much was read
      buffer_lock_state_enum: SILA_BufferLockStatus_Enum { Available_EnumVal, InUseBy_SKYAIFS_EnumVal, InUseBy_MicrokernelIO_EnumVal, PendingDecompression_EnumVal, PendingVerification_EnumVal },
      buffer_access_policy_cap: SILA_CapToken<SILA_BufferAccessPolicy_Record>, // Defines who can get what kind of derived capability to this buffer
      pqc_data_integrity_hash_of_content_opt: SILA_Optional<SILA_PQC_Hash_Record<SHA3_512_SILA_Enum>> // PQC Hash of the current data within the buffer, if computed
    }`
*   **SILA Capability Flow & Management:**
    1.  SKYAIFS (or a dedicated Buffer Manager ASA) requests memory from the Microkernel using `SILA_Microkernel_AllocateMemory_Op` (from `SILA_Microkernel_Internals_Design_V0.1.md`), specifying PQC-protection attributes if needed. The Microkernel returns a `memory_region_main_cap`.
    2.  This capability is stored within the `SILA_Shared_IO_Buffer_Record`.
    3.  When SKYAIFS initiates an I/O operation (e.g., `SKYAIFS_Request_BlockRead_SILA_Call`), it typically creates a derived, restricted `SILA_CapToken` from `memory_region_main_cap` (e.g., granting only write access for a read operation, or read access for a write operation, for a specific size) and passes this derived capability as the `target_buffer_mem_cap` or `source_buffer_mem_cap`.
    4.  The Microkernel's SILA runtime environment verifies the rights and scope of the passed capability before allowing hardware DMA to/from the associated memory region. This is crucial for security and preventing buffer overflows.
    5.  SILA's capability derivation mechanisms (`ADK_Service_Cap.call_Derive_Capability_Restricted(...)`) are used to create these temporary, operation-specific capabilities.

## 4. Atomicity & Consistency Considerations

### 4.1. Challenge: Atomic Metadata Update with Kernel Resource Allocation/Modification
*   **Scenario Example:** Creating a new SKYAIFS file that also requires a unique, kernel-managed security object (e.g., a special PQC key object or a resource quota object) to be associated with it.
    1.  SKYAIFS needs to allocate its internal PQC-signed metadata structures (e.g., `SILA_SKYAIFS_File_Descriptor_Record`).
    2.  SKYAIFS needs to request the Microkernel to create/allocate the associated kernel security object. The capability to this kernel object must then be stored within SKYAIFS's metadata.
*   **Atomicity Requirement:** Both the SKYAIFS metadata commit (PQC-signed, versioned graph update) and the Microkernel resource allocation/update must succeed or fail atomically as a single logical operation. Otherwise, orphaned kernel resources or inconsistent SKYAIFS metadata could result.
*   **Potential SILA-based Approach (Iteration 1 High-Level Thought):**
    *   A **Two-Phase Commit (2PC) SILA Protocol** orchestrated by the initiating SKYAIFS ASA, leveraging SILA's verifiable contract features.
    *   **Phase 1 (Prepare):**
        1.  SKYAIFS ASA: `SILA_Call(Microkernel_TransactionalService_EP_Cap, SILA_PrepareResourceAllocation_Request { desired_resource_spec_cap, transaction_id }) -> SILA_PrepareResource_Response { status, temp_kernel_transaction_cap_opt }`.
        2.  SKYAIFS ASA: Locally prepares its own metadata changes (new SILA graph segments), but does not yet make them globally visible or PQC-sign them as final.
    *   **Phase 2 (Commit/Abort):**
        3.  If local SKYAIFS preparation succeeds AND Microkernel `PrepareResource` was successful:
            SKYAIFS ASA: `SILA_Call(Microkernel_TransactionalService_EP_Cap, SILA_CommitResourceAllocation_Request { kernel_transaction_cap_from_prepare }) -> SILA_Commit_Response { status, final_resource_cap_opt }`.
            If Microkernel commit succeeds, SKYAIFS ASA finalizes and PQC-signs its metadata, making the new file (with reference to `final_resource_cap_opt`) visible.
        4.  If any step in Phase 1 or the Microkernel commit in Phase 2 fails:
            SKYAIFS ASA: `SILA_Call(Microkernel_TransactionalService_EP_Cap, SILA_AbortResourceAllocation_Request { kernel_transaction_cap_from_prepare })`.
            SKYAIFS ASA discards its local metadata changes.
    *   **SILA Requirements:** This necessitates the Microkernel exposing a transactional SILA interface for specific resource types. The `SILA_Module_Contract_Record` for both the SKYAIFS ASA and the Microkernel service would need to specify their roles and obligations in this 2PC protocol. The SILA Verifier could then check for protocol compliance.

## 5. Performance Optimization Considerations (Initial)
*   **SILA IPC Efficiency:** The SILA IPC mechanisms used for high-frequency block I/O must be highly optimized. This might involve the SILA runtime using pre-allocated communication channels or shared memory ring buffers (with access controlled by SILA capabilities) for message passing between SKYAIFS ASAs and the Microkernel Storage Service ASA, minimizing data copying for message headers.
*   **Batching I/O Requests:** SKYAIFS ASAs can batch multiple block read/write requests into a single SILA message (e.g., using `SILA_Array<SILA_BlockRead_SubRequest_Record>`) to reduce IPC overhead per block. The Microkernel would then process these sub-requests, potentially in parallel if targeting different parts of a device or different devices.

## Iteration 1 Conclusion
This first iteration has established the foundational SILA IPC protocols for raw block I/O between SKYAIFS and the Microkernel, including conceptual SILA Record structures for messages and emphasizing the use of SILA V0.2 PQC-aware types and capability management for shared I/O buffers. An initial challenge related to atomicity for combined SKYAIFS metadata and Microkernel resource updates has been identified, with a preliminary SILA-based two-phase commit protocol proposed as a direction for further exploration. Performance considerations for SILA IPC have also been noted.The file `Documentation/Microkernel/SILA_Microkernel_SKYAIFS_Integration_Specification_V0.1.iter1.md` has been successfully created with the specified content.

This completes the simulation of Iteration 1 for the Microkernel-SKYAIFS Integration Specification. The next step is to report this completion.
