# SILA Microkernel & SKYAIFS Integration Specification V0.1

## 1. Introduction
This document details the SILA-based interaction pathways, protocols, and shared data structures between the Skyscope OS Microkernel and SKYAIFS (Skyscope AI Filesystem). The aim is to define precise, secure, and efficient communication for filesystem operations, leveraging **SILA (Sentient Intermediate Language for Agents)**. This builds upon the Stage 2 detailed designs of both components.

## 2. SILA Inter-Process Communication (IPC) Protocols

### 2.1. Raw Block I/O
*   **Request from SKYAIFS to Microkernel (Conceptual SILA Call):**
    `SKYAIFS_RequestBlockRead_SILA_Call(
      target_microkernel_storage_service_ep_cap: SILA_CapToken,
      storage_device_access_cap: SILA_CapToken, // Capability to the specific storage device/partition
      block_id: SILA_LBA_Type, // Logical Block Address
      buffer_for_data_cap: SILA_CapToken // Capability to SKYAIFS-owned memory buffer
    ) -> SILA_Async_Transaction_ID`
    *   The `buffer_for_data_cap` points to a `SILA_IO_Buffer_Record`.
*   **Completion/Status from Microkernel to SKYAIFS (Conceptual SILA Event/Callback):**
    `Microkernel_BlockRead_Complete_SILA_Event(
      target_skyaifs_completion_ep_cap: SILA_CapToken,
      original_transaction_id: SILA_Async_Transaction_ID,
      buffer_with_data_cap: SILA_CapToken, // Same buffer cap, now filled or with error
      read_status: SILA_Storage_Status_Enum { Success, ReadError, DeviceNotFound, PermissionDenied },
      pqc_integrity_hash_of_read_data: SILA_Optional<SILA_PQC_Hash_Record>
    )`
    *   The `buffer_with_data_cap`'s associated `SILA_IO_Buffer_Record` would have its content updated.
    *   Write operations follow a similar pattern (`SKYAIFS_RequestBlockWrite_SILA_Call`, `Microkernel_BlockWrite_Complete_SILA_Event`).
*   **SILA Message Structures:**
    *   Requests and responses will use SILA Records with PQC-aware types for sensitive fields if passing through less trusted domains, though direct kernel<->SKYAIFS IPC is expected to be secure.

### 2.2. Privileged Microkernel Operations for SKYAIFS
*   Requests for mapping device MMIO regions or managing IOMMU (if applicable for advanced storage controllers) will use dedicated, highly privileged SILA IPC channels, requiring SKYAIFS to possess specific `SILA_CapToken`s granted at system initialization.
    *   `SKYAIFS_RequestDeviceMMIO_Map_SILA_Call(mkernel_privileged_ep_cap, device_controller_cap, physical_address, size) -> SILA_Memory_CapToken`

### 2.3. Metadata Operations Requiring Kernel Support
*   For operations requiring atomicity across SKYAIFS metadata and kernel resource state (e.g., allocating a kernel object whose capability is then stored in SKYAIFS metadata), a two-phase commit like protocol expressed in SILA sequence graphs will be used. Phase 1 prepares resources in both, Phase 2 commits. This requires specialized SILA IPC calls.

### 2.4. Fault Reporting
*   **SKYAIFS to Microkernel:**
    `SKYAIFS_ReportStorageFault_SILA_Call(
      mkernel_fault_management_ep_cap: SILA_CapToken,
      storage_device_cap: SILA_CapToken,
      fault_details: SILA_StorageFault_Info_Record // SILA structure with error codes, block numbers etc.
    )`
*   **Microkernel to SKYAIFS:**
    `Microkernel_ReportAsyncDeviceError_SILA_Event(
      skyaifs_device_event_ep_cap: SILA_CapToken,
      storage_device_cap: SILA_CapToken,
      error_info: SILA_AsyncDeviceError_Record
    )`

### 2.5. PQC Key Management Interactions
*   If SKYAIFS needs to derive session keys for specific operations using a kernel-protected master key or a key vault service managed via the kernel:
    `SKYAIFS_RequestKeyDerivation_SILA_Call(kernel_key_vault_ep_cap, context_sila_string, desired_key_spec_sila_enum) -> SILA_EphemeralKey_CapToken`
    *   All such interactions must adhere to the min. 4096-bit PQC security policy defined in Stage 2.

## 3. Shared SILA Data Structures & Capability Management

*   **`SILA_IO_Buffer_Record`:**
    `SILA_IO_Buffer_Record {
      memory_block_cap: SILA_CapToken, // Capability to the actual memory region
      current_size_bytes: SILA_Int,
      max_size_bytes: SILA_Int,
      data_integrity_pqc_hash: SILA_Optional<SILA_PQC_Hash_Record<SHA3_256>>, // PQC hash of the buffer content
      lock_status: SILA_BufferLock_Enum // For concurrency control if buffer is shared
    }`
*   **Capability Establishment:**
    *   SKYAIFS requests memory for I/O buffers from the Microkernel (`SILA_Microkernel_AllocateMemoryForProcess_Op`). The Microkernel returns a `SILA_CapToken` to this memory.
    *   SKYAIFS then passes this memory capability (or a derived capability with restricted rights) to the Microkernel within I/O requests.
    *   Device-specific capabilities are granted to SKYAIFS at initialization or by a trusted system configuration agent.

## 4. Atomicity, Consistency, and Concurrency Control in SILA

*   **Two-Phase Commit (2PC) for Cross-Component Atomicity:**
    *   Conceptual SILA Protocol:
        1.  `SKYAIFS_Prepare_Transaction_SILA_Call(mkernel_tx_coordinator_ep, details_of_op)`
        2.  Microkernel attempts to acquire its local locks/resources, responds `ReadyToCommit` or `Abort`. SKYAIFS does likewise for its state.
        3.  If all parties `ReadyToCommit`, `SKYAIFS_Commit_Transaction_SILA_Call(mkernel_tx_coordinator_ep)`.
        4.  Microkernel and SKYAIFS finalize changes.
    *   This is complex and used sparingly, only for operations where eventual consistency is insufficient.
*   **Concurrency in SILA:** SILA's native concurrency primitives (e.g., guarded commands, transactional memory operations on SILA graphs if available, actor-like message passing) will be used by the Microkernel to handle concurrent requests from SKYAIFS AI bots targeting the same storage resources or kernel objects. Each request carries the bot's SILA capability, allowing fine-grained access control.

## 5. Performance Optimization & Resource Management

*   **Prioritized SILA IPC Channels:** Microkernel SILA IPC endpoints dedicated to storage I/O will have high priority in kernel scheduling to minimize latency.
*   **Zero-Copy Data Transfers (Conceptual):** SILA message structures for I/O will primarily pass `SILA_CapToken`s to shared `SILA_IO_Buffer_Record`s, allowing data to be accessed directly by both SKYAIFS and the Microkernel (and potentially DMA engines) without intermediate copies, where memory protection and SILA capability rules permit.
*   **Resource Allocation:**
    *   CPU time for SKYAIFS AI bots (if they run as distinct SILA processes/TCBs) will be managed by Microkernel scheduling policies (e.g., fair share with optional real-time guarantees for critical recovery bots).
    *   Memory for I/O buffers and SKYAIFS internal caches is allocated via standard SILA Microkernel memory management operations, subject to overall system policies and SKYAIFS's resource quotas.The first document, `SILA_Microkernel_SKYAIFS_Integration_Specification_V0.1.md`, has been created.

Now, I'll proceed to the second document for Stage 3:
**2. AI Pipeline & Microkernel Integration Leads' Output:**
   *   Create: `SILA_Pipeline_Microkernel_Integration_Spec_V0.1.md`
