# SILA OS-Level Containerization Concept V0.1

## 1. Introduction
This document outlines the foundational concepts for OS-level containerization within Skyscope OS. These containers are designed to host **SILA (Sentient Intermediate Language for Agents)** processes and modules, providing isolated, resource-controlled, and securely managed environments. This concept leverages the SILA-based Microkernel primitives and integrates with the AI Pipeline for image management.

## 2. SILA Container Primitives & Objects

### 2.1. `SILA_Container_Descriptor_Record`
A SILA Record structure defining a container's configuration:
`SILA_Container_Descriptor_Record {
  container_id: SILA_UniqueID_String,
  description: SILA_Optional<SILA_String_Record>,
  sila_module_references_list: SILA_Array<SILA_ModuleManifest_Ref_CapToken>, // Capabilities to manifests of modules to be included
  namespace_config_capabilities_array: SILA_Array<SILA_NamespacePolicy_CapToken>, // Capabilities to specific namespace configuration policies
  resource_quota_capabilities_array: SILA_Array<SILA_ResourceQuota_CapToken>, // Capabilities defining resource limits
  initial_entry_point_config: SILA_ContainerEntryPoint_Struct, // Defines the first SILA module/function to run
  pqc_signature_of_descriptor: SILA_PQC_Signature_Record<MLDSA_5>
}`

### 2.2. `SILA_Container_Runtime_State_Struct`
A SILA structure holding the runtime state of an active container instance, managed by the Microkernel:
`SILA_Container_Runtime_State_Struct {
  instance_id: SILA_UniqueID,
  descriptor_cap: SILA_CapToken, // Capability to its descriptor
  status: SILA_ContainerStatus_Enum { Creating, Running, Paused, Stopping, Stopped, Error },
  root_process_caps_array: SILA_Array<SILA_Process_Instance_CapToken>, // Capabilities to root SILA processes within the container
  kernel_resource_handles_list: SILA_Array<SILA_KernelObject_CapToken> // Internal kernel handles for container's isolated resources
}`

### 2.3. Microkernel Operations (Conceptual SILA API)
*   `SILA_Microkernel_CreateContainer_Op(container_descriptor_cap: SILA_CapToken, owner_agent_cap: SILA_CapToken) -> SILA_Result_Record<SILA_Container_Instance_CapToken, SILA_ErrorCode_Enum>`
    *   Microkernel validates descriptor signature, allocates resources, sets up namespaces based on policies, and prepares the container environment.
*   `SILA_Microkernel_DestroyContainer_Op(container_instance_cap: SILA_CapToken, force_flag: SILA_Bool) -> SILA_Status_Enum`
    *   Terminates all SILA processes within, reclaims resources.
*   `SILA_Microkernel_StartContainer_Op(container_instance_cap: SILA_CapToken) -> SILA_Status_Enum`
*   `SILA_Microkernel_StopContainer_Op(container_instance_cap: SILA_CapToken, timeout: SILA_Duration) -> SILA_Status_Enum`
*   `SILA_Microkernel_GetContainerState_Op(container_instance_cap: SILA_CapToken) -> SILA_Container_Runtime_State_Struct`

## 3. Namespace Emulation/Implementation in SILA

Namespaces provide SILA processes within a container with an isolated view of system resources.

*   **PID Namespace:**
    *   The Microkernel maintains a mapping between container-local SILA Process IDs (PIDs) and global system PIDs.
    *   A SILA process within a container, when querying PIDs, sees only those within its own container.
    *   SILA calls related to process management are filtered by the Microkernel based on the calling TCB's container capability.
*   **Mount Namespace (SKYAIFS Integration):**
    *   Each container is associated with a `mount_namespace_policy_cap: SILA_CapToken`.
    *   SKYAIFS, when handling file access requests from a SILA process, consults this capability (passed with the IPC request context) to present a virtualized filesystem view (e.g., a chroot-like environment, specific mount points).
    *   This is managed by SKYAIFS AI bots interpreting the policy referenced by the capability.
*   **Network Namespace:**
    *   A dedicated "Network Namespace SILA Agent" service manages virtual network interfaces (VNICs) for containers.
    *   Each container can be assigned a capability to a VNIC SILA object.
    *   The Microkernel enforces packet filtering rules (defined in SILA policy structures associated with the VNIC cap) for traffic to/from container VNICs.
*   **IPC Namespace:**
    *   SILA endpoint capabilities are inherently local to a process unless explicitly shared.
    *   The Microkernel restricts visibility of named system-wide SILA IPC endpoints based on the requesting container's policy capabilities.
*   **User Namespace (Conceptual):**
    *   Container-local user IDs can be mapped to system-global user IDs. SILA capabilities can be tagged with user/group ownership, and the Microkernel enforces checks based on this mapping for inter-container resource access attempts.

## 4. Resource Control and Management for SILA Containers

*   **CPU Quotas:**
    *   A `SILA_SchedulingParams_Cap` (defining CPU share, period, budget) is associated with the `SILA_Container_Descriptor_Record`.
    *   The Microkernel's SILA scheduler enforces these quotas for all TCBs running as part of SILA processes within that container.
*   **Memory Quotas:**
    *   A `SILA_MemoryQuota_Cap` (defining max memory allocation) is part of the container descriptor.
    *   The Microkernel's `SILA_Microkernel_AllocateMemoryForProcess_Op` will fail if a request from a container process exceeds its remaining quota.
*   **I/O Bandwidth & Device Access:**
    *   Specific device access (e.g., to a raw storage partition for a database container) is granted via explicit SILA capabilities in the container descriptor.
    *   I/O bandwidth quotas can be enforced by I/O scheduler SILA policies within the Microkernel, associated with these device capabilities.
*   **Accounting & Monitoring:**
    *   A "Resource Monitoring SILA Agent" service can query the Microkernel (`SILA_Microkernel_GetContainerResourceUsage_Op`) for actual consumption data and enforce actions (e.g., alerting, terminating container) based on SILA policies.

## 5. Inter-Container Communication (Secure SILA IPC)

*   **Default Deny:** By default, SILA processes in one container cannot communicate with processes in another container or with the host OS services directly.
*   **Explicit Channels:**
    *   A "Container Supervisor SILA Agent" (or a similar trusted entity) can establish explicit SILA IPC channels between containers.
    *   This involves:
        1.  Verifying a request against a system-wide inter-container communication SILA policy.
        2.  Invoking Microkernel operations like `SILA_Microkernel_CreateSharedIPC_EndpointPair_Op(container_A_cap, container_B_cap, channel_policy_cap)` which returns a pair of connected endpoint capabilities.
        3.  Securely distributing these endpoint capabilities to the respective authorized SILA processes within each container.
*   **Mediation:** All such communication is still mediated by the Microkernel's SILA IPC primitives, ensuring capability checks and policy enforcement.

## 6. Conceptual "SILA Container Image" Format

A SILA container image is a PQC-signed package (likely a SILA Record structure itself) managed by the AI Pipeline's module repository.
*   **Contents:**
    *   `image_manifest: SILA_ContainerImageManifest_Record` (see below).
    *   `sila_modules_payload: SILA_Array<SILA_Packaged_Module_Record>` (The actual SILA modules, potentially deeply compressed).
    *   `initial_filesystem_overlay_data: SILA_Optional<SILA_SKYAIFS_DataArchive_Blob>` (Initial data/config files).
    *   `image_pqc_signature: SILA_PQC_Signature_Record<MLDSA_5>`.
*   **`SILA_ContainerImageManifest_Record`:**
    *   `image_name_version: SILA_String_Record`.
    *   `base_os_sila_version_compatibility: SILA_VersionSpec_Record`.
    *   `entry_point_module_name: SILA_String_Record`.
    *   `default_namespace_policy_refs: SILA_Array<SILA_Policy_Identifier>`.
    *   `default_resource_quota_refs: SILA_Array<SILA_Quota_Identifier>`.
    *   `required_host_capabilities_list: SILA_Array<SILA_CapabilityType_Enum>` (e.g., network access, specific device type).

## 7. Security Considerations (Initial)

*   **Strong Isolation:** Relies heavily on Microkernel's SILA capability system and correct namespace emulation to ensure memory, process, and resource isolation.
*   **Attack Surface:** Each container exposes certain SILA IPC endpoints; these must be secured. The "Container Supervisor SILA Agent" is a critical trusted component.
*   **Image Integrity:** PQC signing of SILA container images and their constituent SILA modules is vital. Verification by AI Pipeline and Microkernel before execution.
*   **Kernel Primitives:** Security of containerization depends on the verified correctness of underlying Microkernel SILA operations for resource and namespace management.
*   (Further details to be provided by Security Analyst in `SILA_Integration_Container_Security_Policy_V0.1`).The third document, `SILA_Containerization_Concept_V0.1.md`, has been created.

Next, I will create the fourth document for Stage 3:
**4. Security Analyst Lead's Output:**
   *   Create: `SILA_Integration_Container_Security_Policy_V0.1.md`
