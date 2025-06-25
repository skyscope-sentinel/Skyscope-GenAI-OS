# SILA OS-Level Containerization Concept - V0.1 - Iteration 1

**Based on:** `Documentation/Containerization/SILA_Containerization_Concept_V0.1.md` (initial conceptual placeholder from Stage 3 Task Block 3 assignment)
**Key References:**
*   `Documentation/SILA_Language/SILA_Specification_V0.2.md`
*   `Documentation/Microkernel/SILA_Microkernel_Internals_Design_V0.1.md` (and its V0.2 evolution, assuming parallel refinement)
*   `Documentation/AI_Pipeline_Module_Management/SILA_Pipeline_Microkernel_Integration_Spec_V0.2.md`
*   `Documentation/SKYAIFS_Filesystem/SKYAIFS_Framework_V0.2.md`

**Iteration Focus:** Initial detailing of SILA container descriptor, Microkernel lifecycle APIs, conceptual SILA capability-based approach for PID and Mount namespaces, and attaching resource quotas.

## 1. Introduction
This document begins the detailed conceptualization of OS-level containerization for Skyscope OS. This mechanism is designed to host and isolate **SILA (Sentient Intermediate Language for Agents)** processes and modules, leveraging SILA V0.2 primitives and the capabilities of the SILA-based Microkernel. The overarching goal is to provide strong, verifiable isolation, fine-grained resource control, and secure inter-container communication, all managed and expressed through SILA constructs. PQC security (min 4096-bit equivalent) is a baseline for all relevant data structures and communications.

## 2. SILA Container Primitives & Objects

### 2.1. `SILA_Container_Descriptor_Record` (Conceptual V0.1.iter1 - A SILA Record Type)
This SILA Record structure defines the static configuration and policies for a container instance. It is typically created by a higher-level "Container Management SILA ASA" (which could be part of the AI Pipeline or a dedicated OS service) and then passed to the Microkernel for instantiation.

`SILA_Container_Descriptor_Record {
  container_id_human_readable_str: SILA_String_Record, // For logging and human-readable identification
  unique_template_id_cap: SILA_CapToken<SILA_UniqueID_Object_Type>, // Capability to a unique ID for this descriptor template
  
  // Modules to be loaded into the container's initial environment
  main_sila_module_package_cap: SILA_CapToken<SILA_Packaged_Module_Object_Type>, // Capability to the PQC-signed main SILA module package (binary, manifest) to run
  additional_sila_module_package_caps_array: SILA_Array<SILA_CapToken<SILA_Packaged_Module_Object_Type>>, // Array of capabilities to other SILA modules to be pre-loaded
  
  // Namespace Configuration
  namespace_policy_configuration_caps_map: SILA_Map_Record< // Maps namespace types to their specific policy capabilities
    SILA_NamespaceType_Enum { PID_Namespace_EnumVal, Mount_Namespace_EnumVal, Network_Namespace_EnumVal, IPC_Namespace_EnumVal, User_Namespace_EnumVal }, 
    SILA_CapToken<SILA_NamespaceLifecyclePolicy_Record_Type> // Capability to a policy record defining setup and restrictions for that namespace
  >,
  
  // Resource Allocation Policy
  resource_allocation_policy_bundle_cap: SILA_CapToken<SILA_ContainerResourcePolicy_Bundle_Record_Type>, // Capability to a record bundling all resource quotas
  
  // Initial Capabilities for the Container's Root Process(es)
  initial_capabilities_to_grant_to_root_array: SILA_Array<SILA_CapabilityGrantSpecification_Record_Type>, // Specifies capabilities (e.g., to certain IPC EPs, SKYAIFS paths) for the container's root SILA process(es)
  
  pqc_signature_of_descriptor: SILA_PQC_Signature_Record<MLDSA_5_SILA_Enum> // PQC signature of this descriptor, signed by the authorized Container Management ASA
}`

### 2.2. Microkernel SILA APIs for Container Lifecycle (Conceptual)
These are operations exposed by a "Microkernel_ContainerService_ASA" endpoint.

*   **`SILA_Microkernel_InstantiateContainer_FromDescriptor_Operation`**
    *   **Input SILA Record:** `{ descriptor_to_instantiate_cap: SILA_CapToken<SILA_Container_Descriptor_Record_Type> }`
    *   **Output SILA Record (via Reply):** `SILA_Result_Union<SILA_CapToken<SILA_Container_Runtime_Object_Type>, SILA_Error_Record_Cap>`
    *   **Microkernel Actions (Conceptual SILA Logic):**
        1.  Verify the PQC signature of the `SILA_Container_Descriptor_Record` using a trusted public key for the Container Management ASA.
        2.  Validate the descriptor against system-wide container policies (e.g., resource request sanity checks).
        3.  Create the isolated environment:
            *   Set up namespaces as per `namespace_policy_configuration_caps_map`.
            *   Apply resource limits based on `resource_allocation_policy_bundle_cap`.
        4.  Load the specified SILA modules (main and additional) into this new containerized environment. This involves interacting with the `AI_Pipeline_ModuleManager_ASA` (which may call `DeepCompressionService_ASA`) and then using the Microkernel's own `SILA_Microkernel_CreateProcessFromSILAImage_Operation` for each module, but ensuring the new processes are associated with the container's environment.
        5.  Grant initial capabilities from `initial_capabilities_to_grant_to_root_array` to the container's root process(es).
        6.  Return a `SILA_CapToken<SILA_Container_Runtime_Object_Type>` which is a handle to the live container instance.

*   **`SILA_Microkernel_TerminateContainer_Operation`**
    *   **Input SILA Record:** `{ container_to_terminate_cap: SILA_CapToken<SILA_Container_Runtime_Object_Type>, termination_policy_sila_record_opt: SILA_Optional<SILA_TerminationPolicy_Record> }`
    *   **Output SILA Record (via Reply):** `SILA_OperationStatus_Record`
    *   **Microkernel Actions:** Securely terminate all SILA processes within the container, revoke all capabilities held by or associated with the container, and reclaim all allocated resources.

*   **`SILA_Microkernel_InspectContainerStatus_Operation`**
    *   **Input SILA Record:** `{ container_to_inspect_cap: SILA_CapToken<SILA_Container_Runtime_Object_Type> }`
    *   **Output SILA Record (via Reply):** `SILA_Result_Union<SILA_ContainerLiveStatus_Report_Record_Type, SILA_Error_Record_Cap>`
    *   The `SILA_ContainerLiveStatus_Report_Record` would contain current status (Running, Paused), resource usage metrics (CPU, memory), and a list of active SILA process capabilities within the container.

## 3. Namespace Emulation/Implementation in SILA (Conceptual Examples)

Namespaces provide SILA processes within a container with an isolated view of system resources. This is primarily achieved through Microkernel mediation and SILA's capability system, guided by the `SILA_NamespaceLifecyclePolicy_Record` capabilities provided in the container descriptor.

### 3.1. PID Namespace (Conceptual)
*   **Microkernel Role:** The Microkernel maintains a global, unique set of SILA TCB/Process capabilities. When a SILA process is created *within a container*, the Microkernel associates that process's capability with the container's `SILA_Container_Runtime_Object_Type` capability.
*   **SILA Process View within Container:** When a SILA process inside a container invokes a Microkernel SILA operation like `SILA_Microkernel_ListContainerProcesses_Op(self_container_cap: SILA_CapToken)`, the Microkernel SILA logic filters the global process list. It only returns information (e.g., container-local PIDs, which are just indices or obfuscated IDs, not global capabilities) for processes whose global capabilities are associated with `self_container_cap`. Direct access to other containers' process capabilities is denied by default.
*   **`SILA_NamespaceLifecyclePolicy_Record` for PID Namespace:** This SILA Record (capability referenced in the descriptor) might specify:
    *   `is_pid_namespace_shared_with_host_bool: SILA_Secure_Boolean` (typically false for strong isolation).
    *   `max_processes_in_container_int: SILA_Verifiable_Integer`.

### 3.2. Mount Namespace (Conceptual, integrating with SKYAIFS V0.2)
*   **SKYAIFS Integration:** The `SILA_NamespaceLifecyclePolicy_Record_Type` capability for `Mount_Namespace_EnumVal` (from the container descriptor) effectively points to a SKYAIFS-interpretable "virtual filesystem root" policy. This policy could be a capability to a specific directory in SKYAIFS that serves as the container's root, or a more complex SILA structure defining multiple mount points.
*   **SKYAIFS Role & SILA Mechanics:**
    1.  When a SILA process within a container makes a filesystem-related SILA call to SKYAIFS (e.g., `SKYAIFS_OpenFile_Operation(path_string_record, access_flags_enum)`), the Microkernel, when forwarding this IPC, securely attaches a `SILA_CallingContext_Identifier_Cap` that includes the source `SILA_Container_Runtime_Object_Type` capability.
    2.  SKYAIFS ASAs receive the call and the context capability. They use the container capability to look up its specific `mount_namespace_policy_cap`.
    3.  Based on this policy, SKYAIFS resolves the `path_string_record` relative to the container's virtualized root. For example, a request for `/app/data.bin` might be translated by SKYAIFS to `/srv/containers/container_XYZ/rootfs/app/data.bin` in the global SKYAIFS namespace.
    4.  SKYAIFS uses its own SILA capabilities (to global paths) to access the actual data but only returns data or file handle capabilities to the container process that are valid within its virtualized view and permitted by its specific mount policy (e.g., read-only mounts).

## 4. Resource Control and Management for SILA Containers

### 4.1. Resource Quotas via SILA Capabilities within Container Descriptor
*   The `resource_allocation_policy_bundle_cap: SILA_CapToken<SILA_ContainerResourcePolicy_Bundle_Record_Type>` in the `SILA_Container_Descriptor_Record` points to a SILA structure that aggregates various resource quota policies.
*   **`SILA_ContainerResourcePolicy_Bundle_Record` (Conceptual SILA Record):**
    `{
      bundle_id: SILA_UniqueID_String,
      // Capability to a SILA Record defining CPU scheduling parameters (e.g., share, max percentage, real-time priority if allowed)
      cpu_scheduling_policy_cap: SILA_CapToken<SILA_ProcessSchedulingParameters_Record_Type>, 
      // Capability to a SILA Record defining memory limits
      memory_quota_policy_cap: SILA_CapToken<SILA_MemoryQuotaSettings_Record_Type> { max_physical_memory_bytes_int, max_address_space_bytes_int },
      // Optional: Capability to a policy defining I/O bandwidth limits for specific device types
      io_bandwidth_limits_map_opt: SILA_Optional<SILA_Map_Record<SILA_DeviceType_Enum, SILA_CapToken<SILA_IOBandwidthPolicy_Record_Type>>>,
      // Optional: Array of capabilities explicitly granting or denying access to certain abstract device types
      device_access_control_list_opt: SILA_Optional<SILA_Array<SILA_DeviceAccessPermission_Record_Type>>
    }`
*   **Enforcement by Microkernel (SILA Logic):**
    1.  When the `SILA_Microkernel_InstantiateContainer_FromDescriptor_Operation` is processed, the Microkernel reads the policies referenced by `resource_allocation_policy_bundle_cap`.
    2.  It then configures its internal SILA mechanisms:
        *   The SILA scheduler associates the `cpu_scheduling_policy_cap`'s parameters with all SILA TCBs created for/within that container.
        *   The Microkernel's memory allocation SILA operations (`SILA_Microkernel_AllocateMemoryRegion_Operation`), when invoked by a process within the container (or on its behalf), check against the limits defined in `memory_quota_policy_cap`. Requests exceeding the quota will return a `SILA_ResourceExhausted_Error_Record`.
        *   I/O operations to devices are checked against `device_access_control_list_opt` and bandwidth limits.

## 5. Inter-Container Communication (Secure SILA IPC)
*   **Default Deny:** As per SILA's capability security model, a SILA process in one container cannot inherently address or send SILA IPC messages to a SILA process in another container or to host OS services.
*   **Explicit Channel Creation by "Container Supervisor SILA ASA":**
    1.  An authorized "ContainerSupervisor_ASA" (a privileged OS service ASA) receives a request (via secure SILA IPC) to establish a communication channel between, for example, `ContainerA_ProcessX_EP_Setup_Cap` and `ContainerB_ProcessY_EP_Setup_Cap`. This request would specify the purpose and policy for the channel.
    2.  The Supervisor ASA verifies this request against a system-wide inter-container communication SILA policy.
    3.  If authorized, it invokes a Microkernel SILA operation:
        `SILA_Microkernel_CreateSecure_InterContainer_IPC_Channel_Op(
          source_container_cap: SILA_CapToken<SILA_Container_Runtime_Object_Type>, // For Container A
          target_container_cap: SILA_CapToken<SILA_Container_Runtime_Object_Type>, // For Container B
          channel_policy_cap: SILA_CapToken<SILA_InterContainer_IPC_Policy_Record> // Defines data types allowed, PQC encryption requirements etc.
        ) -> SILA_Result_Union<SILA_IPC_Channel_CapabilityPair_Record, SILA_Error_Record_Cap>`
    4.  The `SILA_IPC_Channel_CapabilityPair_Record` contains two new, unique, and linked `SILA_CapToken<SILA_IPC_Endpoint_Type>` capabilities.
    5.  The Supervisor ASA securely delivers one endpoint capability to the designated process in Container A and the other to the designated process in Container B (e.g., via a pre-established setup IPC channel with each container's initial process).
*   **Microkernel Mediation:** All SILA IPC messages sent via these established channels are still routed and mediated by the Microkernel, which enforces the capability rights and any policies associated with the channel endpoints.

## 6. Conceptual "SILA Container Image" Format
This remains broadly consistent with the Stage 3 Task Block 3 directive, managed by the AI Pipeline and stored (likely PQC-signed and Deep Compressed) in the Module Repository.
*   **SILA Structure (`SILA_ContainerImage_Bundle_Record_Type` - a SILA Record type):**
    `{
      image_metadata_cap: SILA_CapToken<SILA_ContainerImage_Manifest_Record_Type>, // Capability to the main manifest
      sila_module_payloads_map: SILA_Map_Record<SILA_String_Record_ModuleName, SILA_CapToken<SILA_Packaged_Module_Object_Type>>, // Map of module names within image to their packaged SILA objects
      initial_filesystem_overlay_data_blob_opt: SILA_Optional<SILA_CapToken<SILA_PQC_Signed_DataBlob_Object_Type>>, // For initial data/config
      overall_image_pqc_signature: SILA_PQC_Signature_Record<MLDSA_5_SILA_Enum> // PQC Signature covering the manifest and a hash of all payloads
    }`
*   The `SILA_ContainerImage_Manifest_Record_Type` (referenced by `image_metadata_cap`) is essentially the `SILA_Container_Descriptor_Record` adapted for an image template (e.g., resource requests might be relative rather than absolute capabilities). When a container is instantiated from an image, the Container Management ASA resolves these template values into concrete capabilities for the final descriptor.

## Iteration 1 Conclusion
This first iteration has established the foundational SILA Record structures for describing a container (`SILA_Container_Descriptor_Record`) and the primary Microkernel SILA APIs for its lifecycle management (Create, Destroy, Inspect). Initial conceptual approaches for SILA capability-based PID and Mount namespace emulation (the latter integrating with SKYAIFS V0.2) have been outlined. The method for attaching resource quotas to containers via SILA capabilities embedded within the descriptor (specifically the `SILA_ContainerResourcePolicy_Bundle_Record`) has also been introduced. The "SILA Container Image" format has been conceptualized as a PQC-signed SILA Record structure managed by the AI Pipeline. The next iteration will focus on detailing other namespace types (Network, IPC, User), refining inter-container IPC setup mechanisms, and further specifying the image instantiation process.The file `Documentation/Containerization/SILA_Containerization_Concept_V0.1.iter1.md` has been successfully created with the specified content.

This completes the simulation of Iteration 1 for the SILA Containerization Concept. The next step is to report this completion.
