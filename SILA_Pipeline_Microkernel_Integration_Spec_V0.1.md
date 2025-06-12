# SILA Pipeline, Module Management & Microkernel Integration Specification V0.1

## 1. Introduction
This document specifies the integration protocols and SILA-based mechanisms between the Skyscope AI Pipeline (including its Module Manager and interfaces to the Deep Compression service) and the SILA-based Microkernel. The focus is on the secure and efficient loading, management, and runtime support for SILA modules within the Skyscope OS environment. This builds upon Stage 2 detailed designs.

## 2. Detailed SILA Module Loading Protocol

This protocol details the steps from a Module Manager agent requesting a module to the Microkernel preparing it for execution.

**Conceptual SILA Semantic Sequence Graph:**

1.  **Module Request (Module Manager to AI Pipeline Service - SILA IPC):**
    *   `ModuleManager_Agent.SILA_Call(AIPipeline_Service_EP_Cap, SILA_RequestModule_Msg { module_name: SILA_String, version_constraint: SILA_VersionSpec_Record }) -> SILA_Async_Job_ID`

2.  **AI Pipeline Service - Module Retrieval & Decompression:**
    *   Retrieves `SILA_Packaged_Module_Record_Cap` from the Module Repository (internal SILA operation).
    *   Inspects `packaged_module.compression_info`.
    *   If compressed:
        `DeepCompression_Service.SILA_Call(DecompressPayload_Operation, packaged_module.binary_payload_cap, &decompressed_binary_payload_cap)`
    *   Else: `decompressed_binary_payload_cap = packaged_module.binary_payload_cap`.

3.  **AI Pipeline Service - Signature & Manifest Verification:**
    *   Verifies PQC signature of `decompressed_binary_payload_cap` using public key from `packaged_module.manifest.signing_authority_info`.
    *   Verifies PQC signature of `packaged_module.manifest_sila_struct`.
    *   (If verification fails, report error back to ModuleManager_Agent via Job_ID status).

4.  **Module Loading Request (AI Pipeline Service to Microkernel - SILA IPC):**
    *   `AIPipeline_Service.SILA_Call(Microkernel_ModuleLoader_EP_Cap, SILA_LoadModule_Request {
        module_binary_data_cap: SILA_CapToken, // Points to decompressed, verified binary
        module_manifest_cap: SILA_CapToken, // Points to verified manifest SILA structure
        requesting_entity_id: SILA_AgentID_Record // For policy decisions by Microkernel
      }) -> SILA_Async_Job_ID`
    *   The `module_binary_data_cap` grants read-only access to the binary memory.

5.  **Microkernel - `Microkernel_CreateProcessFromSILA_Op` (Internal SILA Logic):**
    *   **Address Space Creation:** Creates a new SILA Address Space object for the module.
    *   **TCB Creation:** Creates initial SILA TCB(s) for the module as specified in the manifest (or a default primary TCB).
    *   **SILA Binary Mapping:** Maps the `module_binary_data_cap` (read-only, execute-only) into the new address space. Maps data sections (read-write) as needed.
    *   **Initial Capability Granting:** Based on `module_manifest_cap.required_capabilities_list` and system policies, mints and grants initial SILA capabilities to the new process's root CSpace (e.g., IPC EPs for standard services, basic memory, fault handler EP).
    *   **Set Entry Point:** Configures the initial TCB's instruction pointer to the SILA binary's entry point specified in the manifest.
    *   Returns a `SILA_Process_Instance_Cap` (capability to the newly created process/main TCB).

6.  **Completion Notification (Microkernel to AI Pipeline Service, then to Module Manager - SILA Events/IPC):**
    *   Microkernel signals success/failure of loading via the Job_ID.
    *   AI Pipeline Service relays this to the original ModuleManager_Agent.

## 3. SILA Runtime Support from Microkernel

The Microkernel provides the following SILA operations/primitives accessible to loaded SILA modules (via pre-granted capabilities):

*   **Dynamic Memory Allocation:**
    *   `SILA_Microkernel_AllocateMemoryForProcess_Op(
        owning_process_cap: SILA_CapToken, // Implicit from caller TCB
        size_bytes: SILA_Int,
        memory_type_enum: SILA_MemoryType { Standard, PQC_Protected_Data, Executable_SILA_Code },
        permissions_enum: SILA_MemoryAccessRights
      ) -> SILA_Result_Record<SILA_Memory_CapToken, SILA_ErrorCode_Enum>`
    *   The returned capability points to a newly allocated memory region.
*   **Inter-Process SILA IPC Channel Setup:**
    *   `SILA_Microkernel_CreateIPC_Endpoint_Op(owning_process_cap: SILA_CapToken) -> SILA_Result_Record<SILA_Endpoint_CapToken, SILA_ErrorCode_Enum>` (creates a new endpoint owned by the process).
    *   `SILA_Microkernel_RequestInterProcessChannel_Op(
        target_process_id_or_service_name: SILA_Identifier,
        channel_policy_sila_struct: SILA_Optional<SILA_IPC_Policy_Record> // e.g., max message size, PQC requirements
      ) -> SILA_Result_Record<SILA_IPC_Channel_CapToken, SILA_ErrorCode_Enum>`
    *   This might involve a system service broker for named services.
*   **Process Termination & Resource Reclamation:**
    *   `SILA_Microkernel_TerminateProcess_Op(process_to_terminate_cap: SILA_CapToken, exit_code: SILA_Int)` (Privileged operation).
    *   `SILA_Microkernel_SelfTerminate_Op(exit_code: SILA_Int)` (Called by a process on itself).
    *   The Microkernel is responsible for revoking all capabilities held by the terminated process and reclaiming its resources (memory, TCBs, etc.). This is a complex SILA graph operation.
*   **Module Status/Performance Queries:**
    *   `SILA_Microkernel_GetProcessResourceUsage_Op(process_cap: SILA_CapToken) -> SILA_ResourceUsage_Record`
    *   `SILA_Microkernel_GetModuleState_Op(process_cap: SILA_CapToken) -> SILA_ModuleState_Enum`

## 4. Dynamic Module Management & Reconfiguration in SILA

*   **Request Unload/Update (AI Pipeline to Microkernel - SILA IPC):**
    *   `AIPipeline_Service.SILA_Call(Microkernel_ModuleLifecycle_EP_Cap, SILA_RequestModuleUnload_Msg { target_process_cap: SILA_CapToken, graceful_shutdown_timeout: SILA_Duration })`
    *   `AIPipeline_Service.SILA_Call(Microkernel_ModuleLifecycle_EP_Cap, SILA_RequestModuleUpdate_Msg { old_process_cap: SILA_CapToken, new_module_binary_cap: SILA_CapToken, new_module_manifest_cap: SILA_CapToken })`
*   **Microkernel Orchestration (SILA Logic):**
    *   **Graceful Shutdown:** Send a `SILA_Shutdown_Notification_Event` to the target module's registered lifecycle endpoint. Wait for acknowledgement or timeout.
    *   **State Transfer (if applicable):** Module may use SILA IPC to transfer state to a replacement version or persistent storage (coordinated by AI Pipeline).
    *   **Resource Revocation:** Microkernel revokes capabilities and reclaims resources of the old module.
    *   **Loading New Version:** Use the module loading protocol (Section 2) for the new version.
*   **Version Compatibility:** Checked by the AI Pipeline using SILA manifest data before initiating an update. The Microkernel enforces that only compatible modules (as per policy) can replace existing ones if critical services are involved.

## 5. Performance and Security Considerations

*   **Predictive Loading/Caching:** The AI Pipeline may predictively decompress and request pre-loading of frequently used SILA modules into a Microkernel-managed cache of verified binaries to speed up subsequent activation. This uses a SILA interface like `SILA_Microkernel_CacheModule_Op(verified_binary_cap, manifest_cap)`.
*   **PQC Signature Verification:** Mandatory for all SILA binaries and manifests by the AI Pipeline Service *before* requesting loading by the Microkernel. The Microkernel may perform an additional fast check (e.g., on a hash) if the binary was in its trusted cache.
*   **Capability Integrity:** The Microkernel ensures that initial capabilities granted to a new SILA module are strictly based on its verified manifest and system policies, preventing unauthorized access.
*   **Isolation:** Loaded SILA modules operate in isolated SILA address spaces with distinct TCBs, as established by the Microkernel.The second document, `SILA_Pipeline_Microkernel_Integration_Spec_V0.1.md`, has been created.

Next, I will create the third document for Stage 3:
**3. Containerization Expert Lead's Output:**
   *   Create: `SILA_Containerization_Concept_V0.1.md`
