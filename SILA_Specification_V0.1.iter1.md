# SILA (Sentient Intermediate Language for Agents) Specification - V0.1 - Iteration 1

**Based on:** `Novel_Language_Specification_V0.1.md`
**Iteration Focus:** Deepening core concepts, PQC explicitness, ADK initial thoughts, policy representation.

## 1. Name
**Proposed Name:** SILA (Sentient Intermediate Language for Agents) - Retained.

## 2. Core Philosophy
Retained from V0.1. Emphasis in this iteration:
*   **Enhanced AI Ergonomics:** Focus on ADK features that simplify complex OS pattern generation by AI agents.
*   **Verifiable PQC Semantics:** Ensuring PQC operations are not just present but their usage context is verifiable.

## 3. Abstract Syntax & Structure

### 3.1. Semantic Graphs
Retained from V0.1.
*   **Elaboration on Edge Types:**
    *   `ControlFlow_Conditional(predicate_graph_ref: SILA_GraphHandle_Cap)`: For conditional branches, where `predicate_graph_ref` points to a SILA subgraph that evaluates to a SecureBoolean.
    *   `DataFlow_Typed(expected_sila_type_descriptor_cap: SILA_CapToken)`: For typed data links, referencing a SILA type descriptor capability.
    *   `CapabilityFlow_Restricted(rights_subset_sila_record: SILA_RightsSpec_Record)`: For passing capabilities with potentially narrowed rights defined in a SILA structure.
    *   `PolicyBinding_Edge(policy_object_cap: SILA_CapToken)`: Links a code segment (a subgraph or node) to an applicable `SILA_ExecutionPolicy_Record` capability.
*   **Elaboration on Node Attributes (Conceptual Additions to Node Definitions):**
    *   `Node_PQC_Context_Attribute { 
          required_security_level: SILA_PQC_SecurityLevel_Enum, // e.g., Level5_4096bit
          associated_key_policy_cap: SILA_Optional<SILA_CapToken>, // Cap to a policy for key derivation/access for this node
          operation_specific_pqc_params: SILA_Optional<SILA_Any_Record> // e.g., specific salt or IV source policy
        }`
    *   `Node_Verification_Annotation { 
          formal_specification_ref_uri: SILA_String_Record, // URI to a formal spec document/section
          proof_status: SILA_ProofState_Enum { Unverified, PartiallyVerified, FullyVerified, VerificationFailed },
          last_verification_job_id: SILA_Optional<SILA_UniqueID>
        }`

### 3.2. Abstract Object Manipulation & Layered Representation
Retained from V0.1. ADK discussion expanded in section 5.

## 4. Key Semantic Concepts

### 4.1. Data Types
*   **Primitive Types:** Retained. `CapabilityToken` is further refined:
    *   `SILA_CapToken<Referenced_SILA_Type_Descriptor_Cap>`: Parameterized by a capability to the SILA type descriptor of the object it references (e.g., `SILA_CapToken<SILA_TCB_Type_Cap>`). This allows stronger type checking for capability usage at the SILA graph level by the Verifier and ADK.
*   **Complex Types:**
    *   **PQC-Aware Structures (Elaboration):**
        *   Conceptual syntax/method for AI agent specification via ADK for creating a PQC-protected data type:
          `ADK_Service_Cap.call_Define_PQC_Struct_Type(
            name: "MyEncryptedData_Type",
            fields_definition: SILA_Map_Record<FieldName_String, SILA_TypeDescriptor_Cap>, // Map of field names to their SILA type descriptor caps
            pqc_policy: SILA_PQC_DataProtectionPolicy_Record {
              encryption_algorithm_choice: MLKEM_1024_SILA_Enum, // Explicit algorithm from a SILA Enum for PQC algos
              key_management_policy_cap: SILA_CapToken, // Capability to a policy defining key generation, retrieval, or derivation logic for this type
              integrity_protection_algorithm_choice: SILA_Optional<SHA3_256_SILA_Enum>, // e.g., for encrypt-then-MAC
              associated_data_fields_for_aead: SILA_Optional<SILA_Array<FieldName_String>>
            }
          ) -> SILA_TypeDescriptor_CapToken`
        *   This implies the ADK translates this high-level AI request into the underlying SILA graph representation of the PQC-aware type. The returned capability points to this new type descriptor.
        *   Instances of such types would implicitly require appropriate key capabilities (scoped by `key_management_policy_cap`) when their PQC operations (encrypt, decrypt) are invoked in a SILA graph.
    *   **Agent-Oriented Records & Immutable Data Structures:** Retained. Further emphasis on ADK providing convenient APIs for constructing and working with immutable collections.
*   **No Nulls:** Retained. `SILA_Optional<Type_Cap>` is the standard way to represent potentially absent values.

### 4.2. Control Flow (Conceptual)
*   **Event-Driven Constructs, State Machines, Concurrent Constructs:** Retained.
*   **Policy-Driven Execution (Elaboration):**
    *   SILA programs can have `SILA_ExecutionPolicy_Record` structures attached to modules, specific graph segments, or even individual capability tokens (via their metadata) using `PolicyBinding_Edge` or similar mechanisms.
    *   `SILA_ExecutionPolicy_Record {
          policy_id: SILA_UniqueID_String,
          description: SILA_Optional<SILA_String_Record>,
          rules: SILA_Array<SILA_PolicyRule_GraphHandle_Cap>, // Each rule is a SILA subgraph that evaluates conditions and actions/assertions
          default_enforcement_level: SILA_PolicyEnforcement_Enum { Strict_FailOnViolation, AuditLog_OnViolation, Advisory_InformAgent },
          pqc_signature_of_policy: SILA_PQC_Signature_Record<MLDSA_5> // Policies must be signed by an authorized policy authority
        }`
    *   The SILA runtime and/or Verifier checks adherence to these policies. Violation of a `Strict_FailOnViolation` policy would typically result in a fault or controlled termination.

### 4.3. Memory Management Model
Retained from V0.1. SILA's interaction with microkernel capabilities for memory regions is key. The ADK will provide abstractions for requesting and using memory capabilities according to defined policies.

### 4.4. AI Integration Features
Retained from V0.1. `Native AI Model Invocation` will use SILA capabilities to reference specific, versioned, and PQC-signed AI models.

## 5. Non-Human Readability & AI Accessibility

### 5.1. Obfuscated Binary Format & Rich Metadata Layer
Retained from V0.1. The PQC-signed metadata layer will include references (e.g., secure hashes or content-addressable URIs) to the formal specifications and policies applicable to the SILA binary.

### 5.2. Agent Development Kit (ADK) - Further Thoughts
*   **Graph Manipulation APIs (Conceptual Additions - as SILA service calls to an ADK service):**
    *   `ADK_Service_Cap.call_Create_ExecutionPolicy(rules_list_caps: SILA_Array<SILA_GraphHandle_Cap>, default_level: SILA_PolicyEnforcement_Enum) -> SILA_ExecutionPolicy_CapToken`
    *   `ADK_Service_Cap.call_Bind_Policy_To_GraphSegment(target_graph_handle_cap: SILA_GraphHandle_Cap, segment_selector_logic_cap: SILA_GraphHandle_Cap, policy_to_bind_cap: SILA_ExecutionPolicy_CapToken)`
    *   (PQC Type Definition API from 4.1. is also part of ADK).
    *   `ADK_Service_Cap.call_Query_Graph_Security_Properties(target_graph_handle_cap: SILA_GraphHandle_Cap, query_policy_cap: SILA_CapToken) -> SILA_SecurityAnalysis_Report_Record` (for AI agent to get security posture overview).
*   **High-Level Pattern Libraries:** The ADK will provide access to a repository of PQC-signed, verifiable SILA graph patterns for common OS tasks (e.g., secure IPC setup, fault-tolerant resource allocation, various synchronization primitives). AI agents can instantiate and customize these patterns via ADK APIs. These patterns are themselves versioned SILA modules.

## 6. Toolchain Concept (AI-Driven)
Retained from V0.1. The SILA Verifier will be critical for checking policy adherence defined in `SILA_ExecutionPolicy_Record` structures, by evaluating the linked `SILA_PolicyRule_GraphHandle_Cap` subgraphs against the target code graph.

## Iteration 1 Conclusion
This iteration focused on adding more explicit detail to PQC type specification and instantiation via the ADK, initial concepts for policy representation and binding, and expanding on ADK graph manipulation APIs relevant to these areas. The goal is to make SILA more concretely specifiable and usable by the AI development team, with a clear path towards verifiability and embedded security. The parameterization of `SILA_CapToken` aims to improve type safety in SILA graph construction.The file `SILA_Specification_V0.1.iter1.md` has been successfully created with the specified enhancements.

This completes the simulation of Iteration 1 for the SILA Specification. The next step is to report this completion.
