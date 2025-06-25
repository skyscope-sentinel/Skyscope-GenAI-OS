# SILA OS-Level Containerization Concept - V0.1 - Iteration 3

**Based on:** `Documentation/Containerization/SILA_Containerization_Concept_V0.1.iter2.md`
**Key References:** 
*   `Documentation/SILA_Language/SILA_Specification_V0.2.md`
*   `Documentation/Microkernel/SILA_Microkernel_Internals_Design_V0.1.md` (V0.2 evolution, assuming parallel refinement)
*   `Documentation/AI_Pipeline_Module_Management/SILA_Pipeline_Microkernel_Integration_Spec_V0.2.md`
*   `Documentation/SKYAIFS_Filesystem/SKYAIFS_Framework_V0.2.md`
*   `Documentation/Security_Policies/Refined_SILA_Security_Policies_Verification_Reqts_V0.1.md` (V0.2 evolution, assuming parallel refinement)

**Iteration Focus:** User namespace emulation, lifecycle management of containerized SILA apps, SILA contracts for container services, formal verification targets, enhanced container image security.

## 1. Introduction
Retained from Iteration 2. This iteration of the SILA Containerization Concept delves into user context management within containers, the lifecycle control of containerized SILA applications, defines formal guarantees via SILA contracts for key containerization services, identifies critical logic for formal verification, and enhances security considerations for SILA container images. All concepts align with SILA V0.2 and PQC security (min 4096-bit).

## 2. SILA Container Primitives & Objects
The refined `SILA_Container_Descriptor_Record` (from Iteration 2, including `container_network_configuration_policy_cap` and `allowed_inter_container_ipc_peers_array`) and the Microkernel SILA APIs (`SILA_Microkernel_InstantiateContainer_FromDescriptor_Operation`, `SILA_Microkernel_TerminateContainer_Operation`, `SILA_Microkernel_InspectContainerStatus_Operation`) remain central.

## 3. Namespace Emulation/Implementation in SILA (Continued)

### 3.1. PID, Mount, Network, IPC Namespaces
Retained and refined from Iteration 2. PID namespace is managed by Microkernel mapping. Mount namespace leverages SKYAIFS V0.2 providing virtualized views based on container's policy capability. Network namespace involves a `System_NetworkNamespaceSupervisor_ASA` and per-container `SILA_VirtualNIC_Object_Type` capabilities. IPC namespace involves container-local registries and Microkernel mediation for global endpoint access.

### 3.2. User Namespace Emulation (SILA - New Detail for Iteration 3)
*   **Global User Identity (Host OS):** Within Skyscope OS, a user interacting with the system (or an AI agent acting on their behalf) is primarily represented by a global `SILA_UserIdentity_Authentication_CapToken`. This capability is managed by a system-wide "UserAuthentication_Service_ASA" and might encapsulate PQC-verified credentials, roles, and permissions at the host OS level.
*   **Container-Local User Context & Mapping:**
    *   The `SILA_Container_Descriptor_Record`'s `namespace_policy_configuration_caps_map` for the `User_Namespace_EnumVal` key points to a `SILA_CapToken<SILA_UserNamespacePolicy_Record_Type>`.
    *   This `SILA_UserNamespacePolicy_Record_Type` defines how the global user identity (of the agent that requested container creation, or an explicitly specified target user identity capability) maps to a user context *within* the SILA container.
    *   **Option 1 (Restricted Identity Passthrough):** The container's root SILA process(es) might inherit a *derived, restricted version* of the global `SILA_UserIdentity_Authentication_CapToken`. When SILA processes within this container interact with Microkernel services or other ASAs like SKYAIFS, this restricted global identity capability is implicitly passed (or explicitly if required by the SILA IPC protocol). These services then use this restricted global identity for their access control decisions. Internally, the container might operate as a single, abstract "container_user_id" (e.g., UID 0 within its world), but external interactions are governed by the restricted global identity.
    *   **Option 2 (Dedicated Container User with Explicit Mapping):** The `SILA_UserNamespacePolicy_Record_Type` specifies a unique, container-local `SILA_LocalUser_Identity_Struct` (e.g., `{ local_uid: 1000, local_gid: 1000, display_name: "app_user" }`). The Microkernel endows the container's initial SILA process(es) with capabilities associated with this local identity for *internal* container operations. For interactions with host OS services (like SKYAIFS global paths, if permitted at all), a special "ContainerUser_Gateway_ASA" (running either in the container with special privileges or as a host service) might be required to explicitly map actions by the `SILA_LocalUser_Identity_Struct` to actions performed by the container's designated global `SILA_UserIdentity_Authentication_CapToken` (if one is associated with the container for external actions). This gateway would be subject to strict SILA policy.
*   **SILA Policy Enforcement by Microkernel & SKYAIFS:**
    *   The `SILA_UserNamespacePolicy_Record_Type` is PQC-signed and its integrity verified by the Microkernel during container instantiation.
    *   The Microkernel tags SILA TCBs running within the container with the appropriate effective user context capability (either the restricted global one or a capability representing the local one).
    *   SKYAIFS (and other resource-managing ASAs) use this user context capability (passed with SILA IPC requests) to make fine-grained access control decisions for file operations or resource access *within the container's mount namespace*.

## 4. Resource Control and Management for SILA Containers
Retained from Iteration 2. Resource quotas are defined by capabilities referenced in `SILA_ContainerResourcePolicy_Bundle_Record_Type` and enforced by the SILA-based Microkernel.

## 5. Secure Inter-Container Communication Setup (SILA Protocol)
Retained and refined from Iteration 2. Communication is mediated by the trusted "System_ContainerInterlink_ASA" and the Microkernel, using PQC-secured SILA IPC channels established via explicit authorization and capability exchange.

## 6. Container Image Instantiation Process (SILA Workflow)
Retained and refined from Iteration 2. The Microkernel instantiates a container from a PQC-signed `SILA_ContainerImage_Bundle_Record_Cap`, verifying all constituent SILA modules and their manifests, and applying container-specific namespaces and resource limits.

## 7. Lifecycle Management of Containerized SILA Applications (New Detail for Iteration 3)

*   **`System_ContainerLifecycleManager_ASA` (CS_ASA - formerly Container Supervisor ASA):** A privileged SILA ASA (likely part of the AI Pipeline's deployment services or a dedicated OS service) responsible for high-level container lifecycle management. It interacts with the Microkernel's container service endpoint.
*   **Starting a Specific SILA Process within an *Existing, Running* Container:**
    *   The `SILA_Container_Descriptor_Record` (or image manifest) defines the main/initial SILA module(s). Additional SILA modules (defined in the image bundle but not started initially, or new modules deployed post-instantiation if policy allows) can be started on demand.
    *   `CS_ASA` sends to `Microkernel_ContainerService_ASA_EP_Cap`:
        `SILA_Container_StartAdditionalProcess_Request_Record {
          target_container_runtime_cap: SILA_CapToken<SILA_Container_Runtime_Object_Type>, // Identifies the running container
          module_to_start_package_cap: SILA_CapToken<SILA_Packaged_Module_Object_Type>, // Capability to the PQC-signed module package
          initial_parameters_for_module_cap_opt: SILA_Optional<SILA_CapToken<SILA_InitialModuleParameters_Record>>,
          reply_to_cs_asa_ep_cap: SILA_CapToken<SILA_IPC_Endpoint_Type>
        }`
    *   The Microkernel (or a "ContainerInternal_Manager_ASA" specific to that container instance, acting on behalf of the Microkernel) uses the standard SILA module loading protocol (decompression if needed via `DeepCompressionService_ASA`, PQC binary verification against manifest, capability endowment) but critically, all operations occur *within the target container's existing namespaces and resource constraints*. A new SILA process/TCB is created *inside* the container.
    *   Response: `SILA_Container_StartAdditionalProcess_Response_Record { ..., new_process_in_container_cap_opt, ... }`.
*   **Stopping/Signaling SILA Processes within a Container:**
    *   `CS_ASA` sends to `Microkernel_ContainerService_ASA_EP_Cap`:
        `SILA_Container_SignalInternalProcess_Request_Record {
          target_container_runtime_cap: SILA_CapToken<SILA_Container_Runtime_Object_Type>,
          target_process_local_id_within_container: SILA_Local_PID_Type, // Container-local PID or unique name tag
          signal_event_sila_record_cap: SILA_CapToken<SILA_Signal_Event_Record_Type> // e.g., GracefulShutdown_Request, Terminate_Immediate, UserDefined_SignalX
        }`
    *   **Microkernel Action (SILA Logic):**
        1.  Verify `CS_ASA` has administrative rights over `target_container_runtime_cap`.
        2.  Translate `target_process_local_id_within_container` to the global `SILA_CapToken<SILA_TCB_Object_Type>` using its internal container process list.
        3.  Securely deliver the `signal_event_sila_record_cap` to the target TCB. This might involve sending a SILA IPC message to a pre-registered signal handling endpoint on that TCB (if the TCB's SILA module supports it via its contract) or directly manipulating the TCB's state (e.g., marking for termination) if the signal is forceful and authorized.

## 8. SILA Contracts for Container Services (Conceptual Excerpts)

### 8.1. Microkernel's `SILA_Microkernel_InstantiateContainer_FromDescriptor_Operation`
*   **`SILA_Module_Contract_Record` Excerpt (for this Microkernel SILA operation's interface):**
    *   `preconditions_graph_cap: PredicateGraph_Cap_Verifying_ContainerDescriptor { // This SILA predicate graph verifies:
          // 1. `descriptor_to_instantiate_cap` is valid, its PQC signature is verified against a trusted Container Management Authority key.
          // 2. All `sila_module_package_ref_caps_array` within the descriptor point to valid, PQC-signed module packages in the repository.
          // 3. All `namespace_policy_configuration_caps_map` reference valid, PQC-signed policy records compatible with Microkernel capabilities.
          // 4. The `resource_allocation_policy_bundle_cap` references a valid, PQC-signed policy that is satisfiable by current system resources.
        }`
    *   `postconditions_graph_cap: PredicateGraph_Cap_Verifying_ContainerInstantiation { // On Success status in response:
          // 1. A new `SILA_Container_Runtime_Object_Type` instance exists and is in a 'Running' or 'Initializing' state.
          // 2. The container provides memory, PID, and other specified namespace isolations according to the descriptor's policies.
          // 3. Resource limits from the descriptor are active and enforced by the Microkernel for this container.
          // 4. The main SILA module(s) specified in the descriptor have been loaded (PQC-verified, decompressed if needed) and started as SILA processes within the container's context.
          // 5. No capabilities are leaked between the container and host, or between this container and others, beyond what's explicitly defined by inter-container IPC setup.
        }`
    *   `temporal_properties_policy_list: [ TemporalPolicy_Cap_Defining_IsolationMaintenance { // LTL Formula: "G (IsContainerActive(C) -> IsIsolationEnforced(C, AllOtherContainers_And_Host))"
            // (Globally, if container C is active, its specified isolation properties are continuously enforced against all other entities)
          } ]`

### 8.2. "System_ContainerInterlink_ASA" (for setting up inter-container SILA IPC)
*   **`SILA_Module_Contract_Record` Excerpt:**
    *   `preconditions_on_ops: {
          "Handle_SILA_RequestInterContainerChannel_Msg": PredicateGraph_Cap_Verifying_InterlinkRequest { // Verifies:
            // 1. The requesting container (source) has a policy in its `SILA_Container_Descriptor_Record` (`allowed_inter_container_ipc_initiation_policy_array`) that permits it to request communication with the `target_service_name` or `target_container_id_hint`.
            // 2. The target container/service exists and is currently registered with the `System_ServiceRegistry_ASA` as exportable.
            // 3. The `client_side_channel_usage_policy_cap` is valid and compatible with the target service's exported interface policy.
          }
        }`
    *   `postconditions_on_ops: {
          "Handle_SILA_RequestInterContainerChannel_Msg": PredicateGraph_Cap_Verifying_InterlinkResponse { // On Success:
            // 1. A pair of new, connected, and suitably restricted SILA endpoint capabilities have been created by the Microkernel.
            // 2. One endpoint capability has been securely delivered to the authorized client ASA in the source container.
            // 3. The other endpoint capability has been securely delivered to the authorized server ASA in the target container.
            // 4. The established channel adheres to the PQC security policies (e.g., encryption) specified in the channel usage policy and/or system defaults.
          }
        }`

## 9. Formal Verification Targets for Containerization Logic (Refined List)

1.  **PID Namespace Isolation Logic (Microkernel SILA):** The SILA logic within the Microkernel that implements PID virtualization for containers. This includes verifying that a SILA process within Container A cannot enumerate, signal, or otherwise interact with SILA processes in Container B or the host OS using PID-based mechanisms, unless explicitly permitted by a shared PID namespace policy (which itself would be a rare, highly audited configuration).
2.  **Capability Confinement within Containers (Microkernel SILA Logic):** The core SILA mechanisms within the Microkernel that ensure capabilities granted to a SILA process running inside a container are confined by that container's boundaries. This means verifying that:
    *   A containerized process cannot use a capability it holds to directly access or manipulate resources outside its container unless that capability explicitly grants cross-container access (e.g., an endpoint capability from the Interlink ASA).
    *   Capabilities cannot be "leaked" or forged from within a container to gain unauthorized access to host resources or other containers. This involves verifying the CSpace setup and capability derivation logic for containerized processes.
3.  **Resource Quota Enforcement by Microkernel (SILA Logic):** The SILA logic in the Microkernel that enforces CPU (via scheduler), memory (via `SILA_Microkernel_AllocateMemoryRegion_Operation` checks), and potentially other I/O resource quotas (as defined in the container's `SILA_ContainerResourcePolicy_Bundle_Record_Type` capability) on all SILA processes within a container. Verification should prove that a container cannot exceed its allocated quotas and thereby cause denial of service to other containers or the core OS.
4.  **Inter-Container IPC Channel Setup (ContainerInterlink_ASA & Microkernel SILA):** The SILA interaction protocol between a client container, the `System_ContainerInterlink_ASA`, a server container, and the Microkernel for establishing an inter-container IPC channel. This includes verifying that all policy checks are correctly performed by the `CI_ASA` and that the Microkernel only creates channel endpoint capabilities with the precise, restricted rights negotiated and authorized by the `CI_ASA`.

## 10. Security of Container Images (Enhanced Considerations from Iteration 2)

*   **Individual SILA Module PQC Signatures within Image Bundle:** Within the `SILA_ContainerImage_Bundle_Record`'s `sila_module_payloads_map`, each `SILA_CapToken<SILA_Packaged_Module_Object_Type>` points to a module package that *must* contain its own independently verifiable PQC signature (for the binary and its manifest). The Microkernel or `AI_Pipeline_ModuleManager_ASA` (during image preparation or container instantiation) must verify these individual module signatures in addition to the signature on the overall image bundle.
*   **Manifest Integrity and Immutability:** The `image_metadata_cap` (pointing to the `SILA_ContainerImage_Manifest_Record_Type`, which is effectively the container's primary configuration) is critical. Its PQC signature guarantees the integrity of the list of SILA modules to be loaded, their versions, the namespace policies, resource quotas, and initial capabilities. This manifest must be treated as immutable once PQC-signed by the "Image Build Service" SILA ASA within the AI Pipeline. Any modification would invalidate the signature.
*   **Secure Build and Curation Process by AI Pipeline:** The AI Pipeline, when constructing a `SILA_ContainerImage_Bundle_Record`, has the responsibility to:
    1.  Ensure it only includes PQC-verified SILA modules sourced from trusted repositories.
    2.  Verify that the combination of modules and their requested capabilities in the manifest adheres to system-wide security policies before bundling.
    3.  Securely PQC-sign the final `SILA_ContainerImage_Bundle_Record` using a trusted "ImageBuildService_SigningKey_CapToken".

## Iteration 3 Conclusion
This iteration has significantly advanced the SILA Containerization Concept by:
1.  Detailing a conceptual design for User Namespace emulation, offering options for restricted identity passthrough or dedicated container users, all managed via SILA policies and capabilities.
2.  Outlining SILA IPC protocols for fine-grained lifecycle management of individual SILA applications (processes/ASAs) *within* running containers, orchestrated by a `System_ContainerLifecycleManager_ASA`.
3.  Providing initial conceptual examples of `SILA_Module_Contract_Record`s for key containerization services like the Microkernel's container creation operation and the "Container Interlink ASA," specifying their formal guarantees.
4.  Identifying and refining a list of critical SILA mechanisms within the containerization framework (PID namespace isolation, capability confinement, resource quota enforcement, and inter-container IPC setup logic) as high-priority targets for formal verification.
5.  Enhancing the security considerations for SILA container images, emphasizing the need for individual PQC signature verification of all constituent SILA modules and the integrity of the overall image manifest.
These developments further solidify the design for a robust, secure, and verifiable OS-level containerization system built entirely within the SILA V0.2 paradigm.The file `Documentation/Containerization/SILA_Containerization_Concept_V0.1.iter3.md` has been successfully created with the specified content.

This completes the simulation of Iteration 3 for the SILA Containerization Concept. The next step is to report this completion.
