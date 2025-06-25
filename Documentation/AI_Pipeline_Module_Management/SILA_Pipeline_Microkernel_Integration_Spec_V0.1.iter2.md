# SILA Pipeline, Module Management & Microkernel Integration Spec - V0.1 - Iteration 2

**Based on:** `Documentation/AI_Pipeline_Module_Management/SILA_Pipeline_Microkernel_Integration_Spec_V0.1.iter1.md`
**Key References:** 
*   `Documentation/SILA_Language/SILA_Specification_V0.2.md`
*   `Documentation/Deep_Compression/Deep_Compression_Framework_V0.2.md`
*   `Documentation/Microkernel/SILA_Microkernel_Internals_Design_V0.1.md` (or its V0.2 evolution)
*   `Documentation/AI_Pipeline_Module_Management/SILA_AI_Pipeline_ADK_Tooling_Concepts_V0.1.md` (or its V0.2 evolution)
*   Relevant sections from `Documentation/OS_Architecture_Synthesis/Skyscope_OS_Integrated_Core_Component_Design_V0.1_SILA_Edition.md`

**Iteration Focus:** Security of module loading (PQC verification, SILA capability endowment), resource management during loading, SILA process initialization details, refined dynamic module update protocol.

## 1. Introduction
Retained from Iteration 1. This iteration focuses on enhancing the security, resource management aspects, and detailing the initialization sequence of the SILA module lifecycle, ensuring tight integration between the AI Pipeline, Deep Compression service, and the Microkernel, all governed by SILA V0.2 principles and PQC security (min 4096-bit).

## 2. Detailed SILA Module Loading Protocol (SILA V0.2)

Refined from Iteration 1.

### 2.1. Security Enhancements in Module Loading

*   **PQC Signature Verification Points (Multi-Stage):**
    1.  **`AI_Pipeline_ModuleManager_ASA` - Initial Package Verification:**
        *   Upon retrieving the `SILA_Packaged_Module_Record_Cap` from its secure repository, the `ModuleManager_ASA` **must** first verify the PQC signature of the *entire package* (e.g., `package_level_pqc_signature` within the record) using a trusted PQC public key corresponding to the repository's signing authority. This ensures the overall package hasn't been tampered with since publication. If invalid, the process aborts, and a `SILA_SecurityAlert_PQC_PackageTamper_Event` is logged and reported.
    2.  **`AI_Pipeline_ModuleManager_ASA` - Manifest Verification:**
        *   It then accesses the `sila_module_manifest_cap` from within the verified package. The `SILA_Module_Manifest_Record` itself contains a PQC signature. The `ModuleManager_ASA` verifies this manifest signature using the public key specified within the manifest's `signing_authority_info` field (which should point to a trusted developer/build agent identity). This step ensures the manifest's integrity and authenticity.
    3.  **`AI_Pipeline_ModuleManager_ASA` - Binary Integrity Pre-Decompression (Optional):**
        *   If the `SILA_Packaged_Module_Record` includes a PQC hash of the (potentially compressed) binary payload, the `ModuleManager_ASA` can verify this hash before sending it for decompression.
    4.  **`Microkernel_ProcessManager_ASA` - SILA Executable Binary Verification (Critical):**
        *   Within its handler for `SILA_Microkernel_CreateProcessFromSILAImage_Request_Record`, *before* any memory mapping or TCB creation for the new process, the `Microkernel_ProcessManager_ASA` **must** perform a final PQC signature verification on the content of the `sila_executable_binary_image_cap` (this is the decompressed binary).
        *   The public key for this verification is typically referenced or directly included (as a PQC-signed `SILA_Certificate_Record_Cap`) within the `sila_module_manifest_cap` that was also part of the request and previously verified by the `ModuleManager_ASA`. The Microkernel must trust the manifest's integrity (due to prior verification) to trust the public key reference it contains.
        *   If this binary signature verification fails, the Microkernel returns `process_creation_status_enum: Binary_SignatureError_EnumVal` in the `SILA_Microkernel_CreateProcess_Response_Record`. This multi-layered PQC verification ensures end-to-end integrity from build to execution.

*   **Secure Initial Capability Endowment by Microkernel (SILA V0.2 Refined):**
    *   The `sila_module_manifest_cap` (pointing to a `SILA_Module_Manifest_Record`) contains a section:
        `initial_capabilities_requested_list: SILA_Array<SILA_RequestedCapability_Descriptor_Record>`.
    *   `SILA_RequestedCapability_Descriptor_Record { // SILA Record for manifest
          capability_name_tag_string: SILA_String_Record, // For debugging/identification
          capability_type_descriptor_cap: SILA_CapToken<SILA_Type_Descriptor_Record>, // Describes the type of capability needed (e.g., IPC Endpoint, Memory Region, Specific Service Interface)
          required_rights_sila_bitmask: SILA_Rights_Bitmask_Type, // Specific rights needed (e.g., Read, Write, Invoke_OperationX)
          is_optional_bool: SILA_Secure_Boolean,
          policy_for_creation_or_scoping_cap_opt: SILA_Optional<SILA_CapToken<SILA_ExecutionPolicy_Record>> // Policy guiding how Microkernel should create/scope this cap
        }`.
    *   The Microkernel's `SILA_Microkernel_CreateProcessFromSILAImage_Operation`, after creating the new process's root CSpace (a SILA object), iterates through `initial_capabilities_requested_list`.
    *   For each request, the Microkernel (based on system-wide security policies, the manifest's verified authenticity, and the optional `policy_for_creation_or_scoping_cap_opt`) performs one of the following SILA operations:
        *   `SILA_Microkernel_MintNewCapability_Op(...)`: For newly created resources like the process's default fault handler IPC endpoint.
        *   `SILA_Microkernel_CopyAndRestrictCapability_Op(...)`: For granting access to existing system services or resources, ensuring the new capability has only the `required_rights_sila_bitmask`.
    *   These newly minted/copied SILA capabilities are installed into the new process's root CSpace at pre-defined slots or as specified in the manifest (e.g., using `capability_name_tag_string` for lookup).
    *   The SILA logic within the Microkernel responsible for capability endowment is a prime candidate for formal verification to prevent incorrect privilege assignment, guided by `SILA_Specification_V0.2.md`'s rules on capability manipulation.

### 2.2. Resource Management During Module Loading (SILA IPC & Microkernel Logic)

*   **Temporary Memory for Decompression (SILA Protocol):**
    1.  `AI_Pipeline_ModuleManager_ASA` determines the potential maximum decompressed size from `package.compression_info` or manifest.
    2.  It sends to `Microkernel_MemoryService_ASA_EP_Cap`:
        `SILA_Microkernel_AllocateSecureMemory_Request_Record {
          request_id: SILA_UniqueID_String,
          requesting_agent_identity_cap: SILA_CapToken, // Authenticates ModuleManager
          size_in_bytes_int: SILA_Verifiable_Integer,
          memory_attributes_sila_record: SILA_MemoryRegionAttributes_Record { // From SILA Spec V0.2
            is_dma_capable_bool: false, 
            required_pqc_protection_level_enum: PQC_Level_None_EnumVal, // Temporary buffer, content will be verified later
            initial_access_rights_for_owner: {ReadWrite_EnumVal},
            expected_lifetime_enum: Ephemeral_ShortLived_EnumVal
          },
          purpose_tag_string_opt: SILA_Optional<SILA_String_Record>{"DeepCompressionService_TempOutputBuffer"}
        }`
    3.  Microkernel responds with `SILA_Microkernel_AllocateSecureMemory_Response_Record { ..., memory_region_cap_opt: SILA_Optional<SILA_CapToken<SILA_Memory_Region_Type>>, ... }`.
    4.  This `memory_region_cap_opt` is then used by `ModuleManager_ASA` to grant the `DeepCompressionService_ASA` temporary write access for the decompressed SILA binary via the `SILA_DeepDecompress_Request_Record`'s `output_buffer_cap` field (newly added field).
*   **Memory for New Process Address Space & Initial Objects:**
    *   This is primarily handled *internally* by the Microkernel's `SILA_Microkernel_CreateProcessFromSILAImage_Operation` logic. The manifest might specify preferred total memory size or attributes for different segments (code, data, stack), which the Microkernel attempts to honor based on system policy and resource availability.
*   **Handling Resource Allocation Failures (SILA Error Propagation):**
    *   If the Microkernel cannot allocate memory at any stage (either for decompression temporary buffer or for the new process itself), it returns a `SILA_Microkernel_AllocateSecureMemory_Response_Record` or `SILA_Microkernel_CreateProcess_Response_Record` with `status: InsufficientResourcesError_EnumVal` (or a more specific error code) and an `error_details_sila_record_opt` capability pointing to a detailed `SILA_ResourceAllocation_Error_Record`.
    *   The `AI_Pipeline_ModuleManager_ASA` receives this error via SILA IPC. Its SILA error handling logic must then:
        1.  Abort the current module loading sequence.
        2.  Securely reclaim any resources it might have already allocated or requested (e.g., inform DeepCompressionService to cancel, release the temp buffer capability back to Microkernel using `SILA_Microkernel_DeallocateMemory_Op`). SILA's capability system ensures only the original requestor/owner can deallocate.
        3.  Propagate the error (as a new SILA error record, wrapping the original) to the initial `Requesting_ASA` that started the module load.

## 3. SILA Process Initialization Details by Microkernel

Elaborating on the internal SILA logic of `SILA_Microkernel_CreateProcessFromSILAImage_Operation` (from Microkernel's perspective):

1.  **Verify PQC Signatures & Manifest Integrity:** (As detailed in Section 2.1). If fails, return error.
2.  **Create Core SILA Process Objects:**
    *   `new_process_object_cap = SILA_InternalKernel_Create_SILA_ProcessObject_Op(...) -> SILA_CapToken<SILA_Process_Object_Type>` (creates the main kernel structure for the process).
    *   `root_cspace_cap = SILA_InternalKernel_Create_CSpace_Op(new_process_object_cap, manifest.cspace_size_hint) -> SILA_CapToken<SILA_CNode_Type>`.
    *   `root_vspace_cap = SILA_InternalKernel_Create_VSpace_Op(new_process_object_cap) -> SILA_CapToken<SILA_AddressSpaceDescriptor_Type>`.
3.  **Map SILA Binary Image into VSpace:**
    *   Iterate through segments defined in `sila_module_manifest_cap` (e.g., code, read-only data, initialized data).
    *   For each segment:
        *   `segment_memory_cap = SILA_InternalKernel_MapMemory_Op(root_vspace_cap, virtual_address_from_manifest, size_from_manifest, source_sila_binary_image_cap_with_offset, permissions_from_manifest_enum)`.
        *   Permissions (e.g., ReadExecute for code, ReadOnly for ROData, ReadWrite for data) are strictly enforced.
4.  **Allocate Stack for Initial TCB:**
    *   `initial_stack_size = manifest.initial_stack_size_int (or default)`.
    *   `initial_stack_mem_cap = SILA_InternalKernel_AllocateProcessMemory_Op(new_process_object_cap, root_vspace_cap, initial_stack_size, {ReadWrite_EnumVal})`. (This maps it too).
5.  **Create and Configure Initial TCB (SILA ASA):**
    *   `initial_tcb_cap = SILA_InternalKernel_Create_TCB_Op(new_process_object_cap, root_vspace_cap, root_cspace_cap, fault_handler_ep_from_manifest_opt_cap)`.
    *   `SILA_InternalKernel_Configure_TCB_State_Op(initial_tcb_cap, SILA_TCB_InitialState_Record {
          entry_point_virtual_address: SILA_VirtualAddress, // From SILA binary metadata within manifest
          stack_pointer_initial_virtual_address: SILA_VirtualAddress, // Top of initial_stack_mem_cap
          // How initial_module_parameters_cap_opt is passed:
          // Option A: Place capability in a well-known register before first execution.
          // Option B: Place capability on the initial stack for the new process to retrieve.
          // This mechanism needs to be standardized by SILA_Specification_V0.2.md for process startup.
          initial_argument_capability_slot_value_opt: SILA_Optional<SILA_CapToken<SILA_InitialModuleParameters_Record>> 
        })`.
6.  **Endow Initial Capabilities into Root CSpace:** (As detailed in Section 2.1, using `SILA_InternalKernel_InstallCapabilityInCSpace_Op`).
7.  **Set Initial TCB State to Ready:** `SILA_InternalKernel_Set_TCB_ExecutionState_Op(initial_tcb_cap, ReadyToRun_SILA_Enum)`.
8.  Return `new_process_instance_cap_opt` (typically the `initial_tcb_cap` or `new_process_object_cap`) in the `SILA_Microkernel_CreateProcess_Response_Record`.

## 4. Refined Dynamic Module Update Protocol (SILA)

Building on Iteration 1's graceful shutdown concept, now incorporating a "Service Registry" SILA ASA:

*   **`ServiceRegistry_ASA`:** A well-known SILA ASA that maintains a PQC-signed, versioned mapping from abstract service names (e.g., "skyscope.services.logging.v1") to active `SILA_CapToken<SILA_IPC_Endpoint_Type>`.
*   **Client ASA Discovery:** Client ASAs, instead of hardcoding service EPs, query `ServiceRegistry_ASA`:
    `ServiceRegistry_ASA_EP_Cap.SILA_Call(SILA_GetServiceEndpoint_Request { service_name: "LoggingService", desired_version_constraint_opt: ">=1.2" }) -> SILA_GetServiceEndpoint_Response { service_ep_cap_opt }`.
*   **Update Process Orchestrated by `AI_Pipeline_ModuleManager_ASA`:**
    1.  Loads `ServiceA_V2_ASA` using normal module loading protocol (Section 2), gets `serviceA_v2_instance_cap` and its primary `serviceA_v2_public_ep_cap`.
    2.  (New Step) `ModuleManager_ASA` informs `ServiceRegistry_ASA` to stage the new version:
        `ServiceRegistry_ASA_EP_Cap.SILA_Call(SILA_StageServiceUpdate_Request { service_name: "ServiceA", new_version_string: "v2.0.0", new_service_ep_cap: serviceA_v2_public_ep_cap })`.
        The `ServiceRegistry_ASA` now knows about V2 but doesn't yet make it the default. New clients requesting "ServiceA" without a specific version might still get V1, or be held, based on registry policy.
    3.  `ModuleManager_ASA` sends `SILA_GracefulShutdown_WithUpdatePending_Request_Event` to `ServiceA_V1_ASA_Instance_Cap`. This event includes the `serviceA_v2_public_ep_cap` if direct state handoff is supported by V1's contract.
    4.  `ServiceA_V1_ASA` completes active requests. If its `SILA_Module_Contract_Record` defines a state migration protocol, it uses SILA IPC to communicate with `serviceA_v2_public_ep_cap` to transfer state. This must be a verifiable and secure exchange.
    5.  `ServiceA_V1_ASA` signals `SILA_ReadyForTermination_Event` to `ModuleManager_ASA`.
    6.  `ModuleManager_ASA` instructs `ServiceRegistry_ASA` to make V2 the primary/default version:
        `ServiceRegistry_ASA_EP_Cap.SILA_Call(SILA_CommitServiceUpdate_Request { service_name: "ServiceA", new_default_version_string: "v2.0.0" })`.
        The `ServiceRegistry_ASA` now directs all new requests for "ServiceA" to `serviceA_v2_public_ep_cap`.
    7.  `ModuleManager_ASA` requests Microkernel to terminate `ServiceA_V1_ASA_Instance_Cap` via `SILA_Microkernel_TerminateProcess_Op`.
*   **SILA Contracts for Updatable Services:** Services designed for dynamic updates must declare this in their `SILA_Module_Manifest_Record` and their `SILA_Module_Contract_Record`. This includes specifying the SILA IPC interfaces they expose for graceful shutdown, state export (if any), and state import (if any). The SILA Verifier can check if a service correctly implements its declared update/migration contract.

## Iteration 2 Conclusion
This iteration has significantly enhanced the security and detail of the SILA module loading protocol by:
1.  Specifying multi-stage PQC signature verification points (package, manifest, binary).
2.  Detailing the secure SILA capability endowment process by the Microkernel based on verified manifests.
3.  Outlining SILA IPC protocols for resource management (e.g., temporary memory for decompression) during module loading, including error propagation.
4.  Providing more conceptual detail on the Microkernel's internal SILA logic for initializing a new SILA process (stack setup, entry point, parameter passing).
5.  Refining the dynamic module update protocol to incorporate a "Service Registry" SILA ASA for managing client redirection and versioning during non-disruptive (graceful) updates.
These refinements provide a more robust foundation for the secure and efficient lifecycle management of SILA modules in Skyscope OS.The file `Documentation/AI_Pipeline_Module_Management/SILA_Pipeline_Microkernel_Integration_Spec_V0.1.iter2.md` has been successfully created with the specified content.

This completes the simulation of Iteration 2 for the Pipeline-Microkernel Integration Specification. The next step is to report this completion.
